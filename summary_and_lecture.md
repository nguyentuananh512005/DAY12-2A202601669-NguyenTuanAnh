# Tóm tắt & Bài giảng: K3 - Ngày 12: Hạ Tầng Cloud & Deployment

Chào Sếp, dưới đây là bài giảng chi tiết và tóm tắt toàn bộ nội dung lý thuyết lẫn thực hành của bài Lab **K3 - Ngày 12: Hạ Tầng Cloud & Deployment**. 

Mục tiêu cốt lõi của bài học này là đưa một AI Agent từ chạy local (`localhost:8000`) lên môi trường Cloud thực tế (địa chỉ công khai), đảm bảo các yếu tố: **Bảo mật**, **Quản lý chi phí**, **Khả năng mở rộng (Scaling)** và **Độ tin cậy (Reliability)**.

---

## MỤC LỤC BÀI GIẢNG
1. [Cấu Trúc Tổng Quan Dự Án](#1-cấu-trúc-tổng-quan-dự-án)
2. [CP1 — 12-Factor Config, Health & Logging](#2-cp1--12-factor-config-health--logging)
3. [CP2 — Docker: Multi-stage & Bảo Mật Image](#3-cp2--docker-multi-stage--bảo-mật-image)
4. [CP3 — API Security: Authentication, Rate Limiting & Cost Guard](#4-cp3--api-security-authentication-rate-limiting--cost-guard)
5. [CP4 — Scaling & Reliability: Stateless, Readiness Probe & Graceful Shutdown](#5-cp4--scaling--reliability-stateless-readiness-probe--graceful-shutdown)
6. [CP5 — Cloud Deployment & Mẹo Thực Hành](#6-cp5--cloud-deployment--mẹo-thực-hành)
7. [Phần Bonus — CI/CD với GitHub Actions](#7-phần-bonus--cicd-với-github-actions)

---

## 1. Cấu Trúc Tổng Quan Dự Án

Dự án này là một API AI Agent viết bằng **FastAPI** và kết nối với **Redis** để quản lý trạng thái.
- **Mã nguồn chính**: Nằm trong thư mục `app/`.
- **Hệ thống Test**: Dùng `pytest` chia làm 5 checkpoint tương ứng với 5 block bài học (`tests/test_cp1.py` đến `test_cp5.py`).
- **Mock LLM**: Tệp `utils/mock_llm.py` giả lập phản hồi của mô hình LLM để học viên thực hành không tốn chi phí API key thật.

---

## 2. CP1 — 12-Factor Config, Health & Logging

### 2.1. Cấu hình theo chuẩn 12-Factor App (`app/config.py`)
- **Triết lý**: *Code giống nhau ở mọi môi trường, cấu hình khác nhau ở từng môi trường*.
- **Thực hành**: Dùng `pydantic-settings` để đọc cấu hình trực tiếp từ biến môi trường.
- **Quy tắc quan trọng**: Các bí mật (như `AGENT_API_KEY`) **không được phép có giá trị mặc định** trong code. Nếu thiếu cấu hình trên Cloud, ứng dụng phải crash lập tức khi khởi động (fail-fast), tránh trường hợp chạy nhưng bị lỗi bảo mật hoặc tốn phí không kiểm soát.

### 2.2. Ghi Log chuẩn JSON (`app/logging_utils.py`)
- **Triết lý**: Log trên môi trường cloud phải được gom tập trung (Log Aggregator). Cách tốt nhất là log ra `stdout` ở định dạng **JSON một dòng** (không thụt lề, không xuống dòng trong một event).
- **Thực hành**: Cài đặt hàm `log_event()` xuất ra chuỗi JSON phẳng. Điều này cho phép dễ dàng truy vấn log, ví dụ: *"User nào đang tiêu tốn nhiều tiền nhất?"*.
- **Lưu ý**: Cần đặt `ensure_ascii=False` khi dump JSON để hỗ trợ hiển thị Tiếng Việt không bị lỗi encode.

### 2.3. Endpoint `/health` (`app/main.py`)
- Chỉ dùng để trả lời câu hỏi: *"Tiến trình này còn sống (liveness) để tiếp tục chạy không?"*.
- **Quy tắc**: Endpoint này **tuyệt đối không được kết nối với Redis hay Database**. Nếu Redis nấc nhẹ, `/health` trả về lỗi khiến hệ thống tự động khởi động lại (restart) container một cách vô ích, dẫn đến lỗi dây chuyền.

---

## 3. CP2 — Docker: Multi-stage & Bảo Mật Image

### 3.1. Build Multi-stage là gì?
Build Multi-stage chia quá trình tạo Docker image thành nhiều giai đoạn (stages):
1. **Stage 1 (Builder)**: Sử dụng base image đầy đủ công cụ để biên dịch thư viện, cài đặt dependencies. Giai đoạn này dung lượng sẽ lớn.
2. **Stage 2 (Runtime)**: Chỉ copy kết quả đã compile từ Stage 1 sang một base image cực nhẹ (`python:3.11-slim`). Bỏ lại các công cụ build dư thừa (compiler, gcc, header files).
- **Kết quả**: Image giảm từ ~1GB xuống dưới **500MB**.

### 3.2. Tối ưu Layer Cache trong Dockerfile
Docker thực hiện cache theo từng dòng lệnh trong `Dockerfile`. Khi có một dòng thay đổi, toàn bộ các cache của các dòng phía sau sẽ bị hủy.
- **Tối ưu**:
  ```dockerfile
  COPY requirements.txt .
  RUN pip install --no-cache-dir -r requirements.txt
  COPY app/ ./app
  ```
  Nhờ đặt lệnh `COPY app/` (thay đổi thường xuyên) phía sau lệnh cài dependencies (ít thay đổi), chúng ta không cần cài lại thư viện mỗi khi sửa code.

### 3.3. Bảo mật container
- **Không chạy container bằng root**: Thêm một user thường (non-root) để chạy ứng dụng:
  ```dockerfile
  RUN useradd --create-home --uid 10001 appuser
  USER appuser
  ```
  Nếu hacker khai thác được lỗ hổng của ứng dụng, chúng chỉ có quyền của user thường, không thể chiếm quyền điều khiển host OS.
- **`.dockerignore`**: Ngăn chặn các file nhạy cảm như `.env`, `.git`, hoặc `.venv` bị đẩy vào Docker image.

---

## 4. CP3 — API Security: Authentication, Rate Limiting & Cost Guard

Để bảo vệ API Agent khỏi việc bị spam và làm cạn kiệt ví tiền gọi LLM, ta áp dụng 3 lớp phòng thủ:

### 4.1. Lớp 1: Authentication (`app/auth.py`)
- Xác thực API key qua Header `X-API-Key`.
- **Toán tử so sánh an toàn**: Dùng `secrets.compare_digest(a, b)` thay vì `a == b`. 
  - *Tại sao?* Phép so sánh `==` sẽ dừng lại ngay khi phát hiện ký tự đầu tiên khác nhau (Timing Attack). Hacker có thể đo thời gian phản hồi để đoán từng ký tự của API Key. `compare_digest` luôn so sánh hết độ dài chuỗi để đảm bảo thời gian xử lý không đổi.

### 4.2. Lớp 2: Rate Limiting (`app/rate_limiter.py`)
- Sử dụng thuật toán **Cửa sổ trượt (Sliding Window)** với Redis Sorted Set (ZSET).
- **Cách hoạt động**:
  - Dùng timestamp làm score.
  - Xóa các request cũ nằm ngoài cửa sổ thời gian (ví dụ: quá 60 giây trước) bằng `ZREMRANGEBYSCORE`.
  - Đếm số lượng request hiện tại trong cửa sổ bằng `ZCARD`.
  - Nếu nhỏ hơn giới hạn, thêm request mới bằng `ZADD` với member duy nhất (`f"{timestamp}:{uuid4()}"`).
- **Lưu ý**: Luôn phải **kiểm tra số lượng trước, ghi nhận sau** để tránh chặn nhầm.

### 4.3. Lớp 3: Cost Guard (`app/cost_guard.py`)
- Giới hạn chi phí tiêu dùng của mỗi User theo tháng.
- Lưu trữ số tiền đã chi tiêu vào Redis theo key: `cost:<user_id>:<YYYY-MM>` và tự động hết hạn (expire) để reset sang tháng mới.
- Chặn gọi LLM nếu chi phí vượt mức ngân sách quy định.

---

## 5. CP4 — Scaling & Reliability: Stateless, Readiness Probe & Graceful Shutdown

### 5.1. Thiết kế Stateless (`app/store.py`)
- Lịch sử hội thoại không được lưu trong bộ nhớ RAM của tiến trình (vì khi scale lên nhiều instance, load balancer sẽ định tuyến request ngẫu nhiên sang các container khác nhau).
- Toàn bộ lịch sử hội thoại phải được lưu vào **Redis** tập trung.
- Sử dụng lệnh `LTRIM` của Redis để giữ số lượng tin nhắn tối đa nhằm giới hạn độ dài prompt gửi lên LLM, tiết kiệm chi phí token.

### 5.2. Liveness vs Readiness Probe (`/ready` endpoint)
- `/ready` (Readiness probe): Kiểm tra xem ứng dụng có đủ điều kiện nhận traffic hay chưa. Endpoint này **PHẢI kiểm tra các dependencies** như kết nối Redis (gọi `ping()`).
- Nếu `/ready` trả về lỗi `503`, Load Balancer sẽ rút container đó ra khỏi danh sách nhận request nhưng **không restart** container. Điều này giúp hệ thống tự phục hồi khi Redis bị quá tải tạm thời.

### 5.3. Graceful Shutdown (`app/lifecycle.py`)
- Khi cập nhật phiên bản mới, Platform gửi tín hiệu `SIGTERM` đến ứng dụng. 
- Ứng dụng phải bắt tín hiệu này, đặt cờ `shutting_down = True`. Lúc này, `/health` và `/ready` lập tức trả về `503` để Load Balancer ngừng gửi request mới.
- Ứng dụng xử lý nốt các request đang dang dở (in-flight requests) rồi mới chính thức dừng tiến trình.
- **Lưu ý**: Nhớ gọi lại handler mặc định của Uvicorn sau khi set cờ để uvicorn tiến hành tắt server đúng quy trình.

---

## 6. CP5 — Cloud Deployment & Mẹo Thực Hành

- Sử dụng các PaaS như **Railway** hoặc **Render**.
- Platform sẽ tự động đọc `Dockerfile` để build và chạy ứng dụng.
- **Quản lý biến môi trường**: Cấu hình các biến `AGENT_API_KEY`, `REDIS_URL` trên dashboard của Cloud. Tuyệt đối không commit các file cấu hình nhạy cảm.

---

## 7. Phần Bonus — CI/CD với GitHub Actions

- Xây dựng file `.github/workflows/ci.yml`.
- Mỗi lần có sự kiện `push` hoặc `pull_request` trên nhánh `main`:
  1. Khởi tạo môi trường ảo Python.
  2. Chạy kiểm thử tự động với `pytest` (loại trừ các test deploy từ xa để tránh tốn thời gian).
  3. Thử build Docker image để đảm bảo không lỗi compiler/dependencies.
  4. Nếu pass tất cả, tự động deploy lên Cloud (CD) thông qua Deploy Token hoặc Webhook.
  5. Chạy Smoke Test (gọi `/health` sau khi deploy xong) để xác nhận dịch vụ chạy tốt.

---

## Lời khuyên của Antigravity khi làm Lab:
1. Sếp hãy chạy lệnh: `pytest tests/ -v -m "not docker"` để xác nhận pytest hoạt động (CP0).
2. Hoàn thành lần lượt từ **CP1** đến **CP5** theo tài liệu `LAB_GUIDE.md`.
3. Kiểm tra tiến độ bằng lệnh: `python grade.py`.
4. Điền đầy đủ thông tin triển khai vào `DEPLOYMENT.md` và trả lời 10 câu hỏi tự lượng giá ở `exercises.md`.

*Chúc Sếp làm bài Lab thật tốt! Có phần nào bị kẹt, hãy nhắn em hỗ trợ code hoặc debug ngay lập tức.*
