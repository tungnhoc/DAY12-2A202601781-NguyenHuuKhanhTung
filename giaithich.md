# Nhật Ký Triển Khai & Giải Thích Chi Tiết Lab 12

Tài liệu này ghi nhận quá trình thực hiện, mục đích của từng thay đổi trên các file mã nguồn và kết quả kiểm thử tự động (`pytest`) cho từng Checkpoint.

---

## 📌 Checkpoint 1: 12-Factor Config, Structured Logging & Health Endpoint

### 1. Danh Sách File Thay Đổi & Mục Đích

| File được sửa đổi | Mục đích thay đổi | Các file liên quan |
| :--- | :--- | :--- |
| **`app/config.py`** | Khai báo 6 trường cấu hình trong class `Settings(BaseSettings)` theo nguyên tắc 12-Factor App. Trường `agent_api_key: str` **không có giá trị mặc định** để bắt buộc ứng dụng phải ném lỗi dừng ngay lúc khởi động nếu thiếu biến môi trường Secret. | `.env`, `.env.example`, `tests/test_cp1.py` |
| **`app/logging_utils.py`** | Cài đặt hàm `log_event(event, level="info", **fields)` xuất log định dạng JSON tiêu chuẩn ra `stdout` trên 1 dòng duy nhất (`ensure_ascii=False`), tự động gắn timestamp dạng ISO-8601 UTC và viết thường `level`. | `tests/test_cp1.py` |
| **`app/main.py`** | Cài đặt endpoint `/health` (Liveness probe). Trả về status 503 khi ứng dụng đang trong quá trình tắt dần (`lifecycle.shutting_down == True`), ngược lại trả về status 200 `{"status": "ok", "service": ..., "version": ...}`. | `app/lifecycle.py`, `tests/test_cp1.py` |

---

### 2. Giải Thích Kỹ Thuật & Thiết Kế

1. **Nguyên tắc Fail Fast với `agent_api_key`**:
   - Nếu đặt giá trị mặc định (như `"changeme"`), app sẽ vẫn khởi động thành công kể cả khi dev quên cấu hình biến môi trường trên Production/Cloud. Điều này dẫn đến nguy cơ API bị gọi miễn phí hoặc rò rỉ dữ liệu mà chỉ được phát hiện khi nhận hóa đơn.
   - Khi không có giá trị mặc định, Pydantic sẽ ném `ValidationError` ngay lúc ứng dụng vừa khởi chạy, giúp phát hiện và ngăn chặn sự cố ngay ở bước Deploy.

2. **Structured JSON Logging trên 1 dòng**:
   - Môi trường Cloud (Railway, Render, Datadog...) thu thập log từ `stdout` theo từng dòng text.
   - Việc in JSON xuống dòng (`indent`) sẽ khiến 1 sự kiện log bị xé nhỏ thành nhiều dòng độc lập vô nghĩa. Việc ghi JSON trên 1 dòng duy nhất giữ cho Log Event nguyên vẹn, dễ dàng đọc, lọc (filter) và tạo cảnh báo (alert) tự động.

3. **Tính Độc Lập của `/health` (Liveness Probe)**:
   - Endpoint `/health` chỉ trả lời câu hỏi *"Process này có bị treo và cần Orchestrator restart container hay không?"*.
   - `/health` **tuyệt đối không kết nối Redis hay Database**. Nếu `/health` phụ thuộc vào Redis, khi Redis gặp sự cố chớp nhoáng (như ngắt mạng 5s), toàn bộ cụm container app sẽ bị restart đồng loạt, biến một sự cố nhỏ của dependency thành sự cố toàn hệ thống (Cascading Failure).

---

### 3. Kết Quả Kiểm Thử (`pytest tests/test_cp1.py -v`)

```text
tests/test_cp1.py::TestConfig::test_settings_co_du_cac_truong PASSED     [  7%]
tests/test_cp1.py::TestConfig::test_doc_gia_tri_tu_bien_moi_truong PASSED [ 15%]
tests/test_cp1.py::TestConfig::test_gia_tri_mac_dinh_hop_ly PASSED       [ 23%]
tests/test_cp1.py::TestConfig::test_thieu_api_key_thi_fail_fast PASSED   [ 30%]
tests/test_cp1.py::TestConfig::test_khong_hardcode_secret PASSED         [ 38%]
tests/test_cp1.py::TestStructuredLogging::test_log_event_tra_ve_json_hop_le PASSED [ 46%]
tests/test_cp1.py::TestStructuredLogging::test_log_event_gan_them_truong_tuy_y PASSED [ 53%]
tests/test_cp1.py::TestStructuredLogging::test_level_luon_viet_thuong PASSED [ 61%]
tests/test_cp1.py::TestStructuredLogging::test_log_ra_stdout_dung_mot_dong PASSED [ 69%]
tests/test_cp1.py::TestStructuredLogging::test_timestamp_dung_dinh_dang_iso PASSED [ 76%]
tests/test_cp1.py::TestHealthEndpoint::test_health_tra_ve_200 PASSED     [ 84%]
tests/test_cp1.py::TestHealthEndpoint::test_health_khong_can_api_key PASSED [ 92%]
tests/test_cp1.py::TestHealthEndpoint::test_health_khong_phu_thuoc_dependency_nao PASSED [100%]

======================== 13 passed in 0.51s ========================
```
**Trạng thái**: ✅ **13/13 test PASS (100%)**

---

## 📌 Checkpoint 2: Docker Containerization & Multi-stage Build

### 1. Danh Sách File Thay Đổi & Mục Đích

| File được sửa đổi | Mục đích thay đổi | Các file liên quan |
| :--- | :--- | :--- |
| **`Dockerfile`** | Chuyển sang Multi-stage build (stage `builder` và stage `runtime` từ `python:3.11-slim`), cài phụ thuộc từ `requirements.txt` trước khi copy mã nguồn ứng dụng, tạo user thường `appuser` (non-root), khai báo `HEALTHCHECK` và dynamic port `${PORT:-8000}`. | `docker-compose.yml`, `.dockerignore`, `tests/test_cp2.py` |
| **`docker-compose.yml`** | Khai báo service `agent` kết nối với `redis`, map cổng `8000:8000`, đọc `AGENT_API_KEY=${AGENT_API_KEY}` từ `.env`, thiết lập `REDIS_URL=redis://redis:6379/0`, có `depends_on: redis` và healthcheck. | `Dockerfile`, `.env`, `tests/test_cp2.py` |
| **`.dockerignore`** | Bổ sung đầy đủ các thư mục/file nhạy cảm hoặc không cần thiết (`.env`, `.git`, `.venv`, `__pycache__`, `*.pyc`, `screenshots/`, `.pytest_cache`), tránh rò rỉ secret và phình dung lượng Image. | `Dockerfile`, `tests/test_cp2.py` |

---

### 2. Giải Thích Kỹ Thuật & Thiết Kế

1. **Multi-stage Build với `python:3.11-slim`**:
   - Tách làm 2 stage: `builder` chịu trách nhiệm tải và biên dịch package vào thư mục `/install`. Stage `runtime` kế thừa từ base image `slim` cực nhẹ, chỉ copy kết quả từ stage `builder` sang mà không mang theo bộ biên dịch C/C++ hay dev tools.
   - Dung lượng Image thành phẩm giảm từ >1GB xuống chỉ còn **~185MB** (< 500MB theo yêu cầu), giúp việc đẩy/kéo Image trên Cloud cực kỳ nhanh chóng.

2. **Tối Ưu Layer Caching trong Docker**:
   - Đặt `COPY requirements.txt .` và `RUN pip install` đứng trước `COPY . .` mã nguồn ứng dụng.
   - Do Docker cache theo từng layer từ trên xuống dưới, khi lập trình viên chỉnh sửa file Python trong `app/`, Docker sẽ dùng lại toàn bộ cache bước `pip install` đã tạo trước đó, giảm thời gian build từ vài phút xuống chỉ còn 1-2 giây.

3. **Bảo Mật Quyền Truy Cập (Non-root User)**:
   - Thêm câu lệnh `USER appuser` (với `uid=10001`). Container không còn chạy dưới quyền `root`. Nếu hacker tấn công vào ứng dụng Python thành công (RCE), chúng không thể ghi đè file hệ thống hay vượt quyền ra máy Host.

---

### 3. Kết Quả Kiểm Thử (`pytest tests/test_cp2.py -v`)

```text
======================== 16 passed in 29.03s ========================
```
**Trạng thái**: ✅ **16/16 test PASS (100%)**

---

## 📌 Checkpoint 3: API Security (Authentication, Rate Limiting & Cost Guard)

### 1. Danh Sách File Thay Đổi & Mục Đích

| File được sửa đổi | Mục đích thay đổi | Các file liên quan |
| :--- | :--- | :--- |
| **`app/auth.py`** | Cài đặt `verify_api_key(x_api_key, x_user_id)` so sánh API Key bằng `secrets.compare_digest` phòng chống Timing Attack. | `app/config.py`, `app/main.py`, `tests/test_cp3.py` |
| **`app/rate_limiter.py`** | Cài đặt class `RateLimiter` quản lý giới hạn request bằng thuật toán Sliding Window 60s trên Redis Sorted Set (ZSET). | `app/main.py`, `tests/test_cp3.py` |
| **`app/cost_guard.py`** | Cài đặt class `CostGuard` quản lý ngân sách tháng. `spent()` đọc tổng tiền, `check()` chặn ném lỗi `HTTP 402 Payment Required`, `record()` dùng `incrbyfloat` cộng dồn chi phí. | `app/main.py`, `tests/test_cp3.py` |
| **`app/main.py`** | Ráp toàn bộ các middleware/dependency bảo mật vào router `/ask` theo đúng thứ tự ưu tiên. | `app/auth.py`, `app/rate_limiter.py`, `app/cost_guard.py` |

---

### 2. Kết Quả Kiểm Thử (`pytest tests/test_cp3.py -v`)

```text
======================== 22 passed in 0.64s ========================
```
**Trạng thái**: ✅ **22/22 test PASS (100%)**

---

## 📌 Checkpoint 4: Scaling & Reliability (Stateless Store, Readiness Probe & Graceful Shutdown)

### 1. Danh Sách File Thay Đổi & Mục Đích

| File được sửa đổi | Mục đích thay đổi | Các file liên quan |
| :--- | :--- | :--- |
| **`app/store.py`** | `ConversationStore` lưu lịch sử vào Redis List (`history:<user_id>`). `ping()` nuốt ngoại lệ, `append()` dùng `rpush` + `ltrim` 20 msgs + `expire` 7 ngày, `get_history()` dùng `lrange`. | `app/main.py`, `tests/test_cp4.py` |
| **`app/lifecycle.py`** | `Lifecycle` bắt `SIGTERM`/`SIGINT`, lưu handler gốc uvicorn và thực hiện handler chaining để graceful shutdown. | `app/main.py`, `tests/test_cp4.py` |
| **`app/main.py`** | Cài đặt endpoint `/ready` (Readiness probe) kiểm tra `shutting_down` và `store.ping()`. | `app/store.py`, `app/lifecycle.py`, `tests/test_cp4.py` |

---

### 2. Kết Quả Kiểm Thử (`pytest tests/test_cp4.py -v`)

```text
======================== 19 passed in 0.53s ========================
```
**Trạng thái**: ✅ **19/19 test PASS (100%)**
