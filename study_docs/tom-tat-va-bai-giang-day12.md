# Bài giảng chuyên sâu: Hạ tầng Cloud & Deployment cho AI Agent

Chào Sếp, dưới đây là tài liệu tổng hợp và giảng giải chi tiết về các khái niệm cốt lõi của bài học **K3 Ngày 12: Hạ Tầng Cloud & Deployment**. Tài liệu này được biên soạn kỹ lưỡng để Sếp nắm bắt nhanh chóng và lưu trữ phục vụ cho việc ôn tập hoặc import vào NotebookLM.

---

## I. Cấu hình theo chuẩn 12-Factor App & Pydantic Settings
### 1. Triết lý 12-Factor Config
* **Nguyên tắc**: *Code giống nhau ở mọi môi trường, cấu hình khác nhau ở từng môi trường*.
* **Vấn đề thực tế**: Trong phát triển phần mềm, việc "hardcode" (viết cứng) thông tin cấu hình như API keys, Database URLs, port chạy ứng dụng vào mã nguồn là cực kỳ nguy hiểm. Khi chuyển mã nguồn từ máy cá nhân (localhost) lên môi trường kiểm thử (staging) hoặc môi trường chạy thật (production), bạn sẽ buộc phải sửa code. Điều này dễ dẫn đến rò rỉ các API key nhạy cảm lên GitHub.
* **Giải pháp**: Tách toàn bộ cấu hình ra ngoài code và truyền vào thông qua **Biến môi trường (Environment Variables)**. Cùng một file build (Docker image) sẽ được tái sử dụng ở mọi nơi mà không cần build lại, chỉ thay đổi biến môi trường đi kèm.

### 2. Pydantic-Settings & Fail-Fast
Trong Python, chúng ta sử dụng thư viện `pydantic-settings` để tự động đọc các biến môi trường vào các thuộc tính của Class `Settings`.
* **Fail-Fast (Chết sớm để an toàn)**: Trong `app/config.py`, biến `agent_api_key` **không được phép có giá trị mặc định**.
  ```python
  class Settings(BaseSettings):
      port: int = 8000                  # Có giá trị mặc định
      agent_api_key: str                # BẮT BUỘC - KHÔNG có mặc định!
      redis_url: str = "redis://..."
  ```
  Nếu Sếp deploy ứng dụng lên Cloud mà quên cấu hình biến `AGENT_API_KEY`, ứng dụng sẽ crash lập tức khi vừa khởi động (`ValidationError`). Điều này giúp Sếp phát hiện lỗi cấu hình ngay lập tức thay vì để app chạy bình thường nhưng sử dụng một API key mặc định, dẫn đến lỗi bảo mật hoặc tốn phí không kiểm soát do người lạ gọi API.

---

## II. Structured Logging (Ghi log có cấu trúc)
### 1. Phân biệt log máy đọc và log người đọc
* **Log người đọc (Human-readable)**:
  `print("User sv01 vừa hỏi GPT-4 hết 0.005 USD")`
  Log này rất dễ đọc trên terminal cá nhân. Nhưng trên Cloud (chạy hàng chục container song song), hàng triệu dòng log như thế này đổ về một nơi tập trung (Log Aggregator như Grafana Loki, Datadog). Máy tính không thể parse được câu chữ tự do đó để thống kê.
* **Log máy đọc (Machine-readable - Structured Log)**:
  In ra console (`stdout`) một chuỗi **JSON phẳng trên một dòng duy nhất**:
  ```json
  {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T09:30:00.000Z", "user_id": "sv01", "cost_usd": 0.005}
  ```
* **Tại sao phải là một dòng duy nhất (không indent)?**
  Các hệ thống Cloud thu thập log theo từng dòng. Nếu Sếp dùng `json.dumps(..., indent=4)`, một log event sẽ bị bẻ thành nhiều dòng, hệ thống thu thập log sẽ hiểu lầm đó là các log độc lập, làm vỡ dữ liệu log.
* **Thực hành**: Hàm `log_event()` trong `app/logging_utils.py` phải tạo ra flat JSON, ghi đè `ensure_ascii=False` để hiển thị tiếng Việt chính xác và in thẳng ra `sys.stdout`.

---

## III. API Security nâng cao
Để bảo vệ API Agent trước các nguy cơ spam và làm cạn kiệt ngân sách gọi LLM, ta xây dựng 3 tầng bảo vệ:

### 1. Authentication & Chống Timing Attack
* **Xác thực**: Kiểm tra token trong Header `X-API-Key`.
* **Timing Attack (Tấn công đo thời gian)**: 
  Khi Sếp so sánh chuỗi bằng toán tử `==` (ví dụ: `x_api_key == correct_key`), Python so sánh từng ký tự từ trái qua phải. Ngay khi thấy một ký tự không khớp, nó sẽ dừng và trả về `False`. 
  Hacker có thể gửi API key giả và đo thời gian phản hồi siêu nhỏ (đến mili-giây). Nếu ký tự đầu đúng, phép so sánh chạy lâu hơn một chút so với khi ký tự đầu sai. Bằng cách thử nhiều lần, hacker có thể dò ra từng ký tự của API Key.
* **Giải pháp**: Dùng `secrets.compare_digest(a, b)` trong `app/auth.py`. Hàm này so sánh toàn bộ chuỗi ở thời gian không đổi (constant-time), dù sai ở ký tự đầu hay cuối thì thời gian xử lý vẫn giống hệt nhau, loại bỏ hoàn toàn nguy cơ Timing Attack.

### 2. Rate Limiting cửa sổ trượt (Sliding Window) với Redis
* **Vấn đề của Fixed Window (Cửa sổ cố định)**:
  Nếu giới hạn 10 request/phút theo giờ tròn (từ 10:00 đến 10:01). Hacker có thể gửi 10 request vào giây `10:00:59` and gửi tiếp 10 request nữa vào giây `10:01:01`. Tổng cộng 20 request trong vòng 2 giây mà vẫn đúng luật cửa sổ cố định, gây quá tải hệ thống.
* **Giải pháp Sliding Window (Cửa sổ trượt)**:
  Giới hạn 10 request trong 60 giây gần nhất tính từ thời điểm hiện tại.
* **Triển khai trong Redis**:
  Dùng cấu trúc dữ liệu **Sorted Set (ZSET)**:
  - **Score**: Là timestamp hiện tại (giây).
  - **Member**: Phải là một chuỗi duy nhất, ví dụ `f"{timestamp}:{uuid}"` (nếu chỉ lưu timestamp, hai request gửi cùng một phần nghìn giây sẽ bị ghi đè lẫn nhau, dẫn đến đếm thiếu).
  - **Các bước thực hiện**:
    1. Dọn dẹp các bản ghi cũ nằm ngoài cửa sổ: `ZREMRANGEBYSCORE key 0 (now - 60)`.
    2. Đếm số request hiện tại trong ZSET: `ZCARD key`.
    3. Nếu vượt quá limit -> Báo lỗi `429 Too Many Requests`.
    4. Nếu chưa vượt -> Thêm request mới: `ZADD key now "now:uuid"`, rồi set expire cho key để tự động dọn dẹp sau khi hết hạn.
  *Lưu ý quan trọng*: Phải **kiểm tra trước, ghi nhận sau**. Nếu ghi nhận request trước rồi mới đếm, hệ thống sẽ chặn nhầm request ngay tại ngưỡng giới hạn.

### 3. Cost Guard (Giới hạn chi tiêu)
* Rate limit chỉ giới hạn số lượng request, không giới hạn kích thước dữ liệu. Một user chỉ gửi 1 request/phút nhưng request đó chứa 1 triệu tokens vẫn có thể làm bay sạch tiền trong tài khoản OpenAI của bạn.
* **Giải pháp**: Cost Guard lưu trữ tổng số tiền user đã tiêu trong tháng vào Redis dưới key `cost:<user_id>:<YYYY-MM>`. 
* Khi nhận request `/ask`, hệ thống tính toán chi phí ước lượng của câu hỏi. Nếu `tiền đã tiêu + chi phí ước lượng > ngân sách tháng` -> Từ chối xử lý và trả về mã lỗi `402 Payment Required` (Đúng ngữ nghĩa cho việc hết ngân sách).
* Nếu hợp lệ -> Gọi LLM, sau đó cộng dồn chi phí thực tế trả về vào Redis bằng `incrbyfloat()`. Set TTL cho key là 40 ngày để đối soát và tự động dọn dẹp.

---

## IV. Độ tin cậy & Khả năng mở rộng (Scaling & Reliability)
### 1. Thiết kế Stateless (Không lưu trạng thái tại local)
* Khi deploy ứng dụng lên Cloud, hệ thống Load Balancer sẽ phân phối các request ngẫu nhiên tới các container khác nhau.
* Nếu lịch sử hội thoại lưu trong bộ nhớ RAM của container A, khi user gửi câu hỏi thứ 2 và request rơi vào container B, container B sẽ không biết gì về ngữ cảnh trước đó. Ngoài ra, container có thể bị restart hoặc tắt đi bất cứ lúc nào khi scale-down.
* **Giải pháp**: Lưu toàn bộ lịch sử hội thoại vào **Redis tập trung**. Các container đều không giữ trạng thái (Stateless), chỉ đóng vai trò xử lý logic và đọc/ghi data từ Redis.
* **Tối ưu Prompt Context**: Sử dụng lệnh `LTRIM` của Redis để chỉ giữ lại tối đa `HISTORY_MAX_MESSAGES = 20` tin nhắn gần nhất. Việc này ngăn prompt phình to vô hạn, giúp tiết kiệm chi phí token và tránh vượt quá context window của mô hình.

### 2. Phân biệt Liveness Probe (/health) và Readiness Probe (/ready)
Các hệ thống như Docker Compose, Kubernetes, Railway sử dụng các Probe để theo dõi sức khỏe ứng dụng:
* **Liveness Probe (/health)**:
  - *Mục đích*: Kiểm tra xem ứng dụng còn sống hay đã bị treo cứng (deadlock).
  - *Xử lý khi lỗi*: Hệ thống sẽ **kill container và khởi động lại**.
  - *Quy tắc thiết kế*: Endpoint này phải cực kỳ nhẹ, **tuyệt đối không kết nối với DB hay Redis**. Nếu Redis nấc nhẹ và `/health` trả về lỗi, hệ thống sẽ restart container vô ích, làm tăng thời gian downtime và gây lỗi lan truyền. Nó chỉ kiểm tra cờ tắt máy của chính container đó.
* **Readiness Probe (/ready)**:
  - *Mục đích*: Kiểm tra xem ứng dụng đã sẵn sàng nhận traffic phục vụ khách hàng chưa.
  - *Xử lý khi lỗi*: Hệ thống sẽ **rút container ra khỏi Load Balancer** để không nhận traffic nữa, **nhưng KHÔNG restart**.
  - *Quy tắc thiết kế*: Endpoint này **phải ping thử Redis/Database** để đảm bảo kết nối thông suốt. Nếu kết nối Redis bị mất tạm thời, container sẽ ngừng nhận request và đợi kết nối tự phục hồi, tránh lỗi 500 cho người dùng.

### 3. Graceful Shutdown (Tắt máy êm đẹp)
* Khi cập nhật phiên bản mới, Platform gửi tín hiệu **SIGTERM** yêu cầu container dừng lại. Mặc định nếu không xử lý, container sẽ bị ngắt đột ngột, làm đứt các request đang xử lý (In-flight requests), người dùng sẽ thấy lỗi `502 Bad Gateway`.
* **Giải pháp**:
  1. Đăng ký hàm bắt tín hiệu `signal.SIGTERM` và `signal.SIGINT` trong `app/lifecycle.py`.
  2. Khi nhận tín hiệu, bật cờ `shutting_down = True`.
  3. Lúc này, các endpoint `/health` và `/ready` lập tức trả về lỗi `503 Service Unavailable`. Load Balancer sẽ lập tức ngừng đẩy request mới vào container này.
  4. Container xử lý nốt các request đang chạy dở dang.
  5. Gọi lại handler tắt máy mặc định của uvicorn để server tắt hẳn một cách êm đẹp.

---

## V. Docker Multi-stage & Bảo mật Image
### 1. Build Multi-stage là gì?
Build Docker image thông thường sẽ kéo theo toàn bộ trình biên dịch (gcc, make, python-dev) làm dung lượng image phình to (lên tới 1GB).
* **Multi-stage** chia quá trình build thành 2 giai đoạn:
  - **Stage 1 (Builder)**: Dùng base image đầy đủ để cài đặt thư viện, biên dịch các C-extensions. Dung lượng giai đoạn này lớn.
  - **Stage 2 (Runner)**: Dùng base image siêu nhẹ (`python:3.11-slim`), chỉ copy các thư viện đã cài đặt thành công từ Stage 1 sang. Bỏ lại toàn bộ compiler dư thừa.
* **Kết quả**: Dung lượng image giảm đáng kể (thường dưới 500MB), giúp deploy nhanh hơn và giảm diện tích bị tấn công (attack surface) do loại bỏ công cụ build thừa.

### 2. Bảo mật container bằng Non-root User
Mặc định, Docker chạy container dưới quyền `root`. Nếu ứng dụng có lỗ hổng bảo mật (ví dụ: Remote Code Execution), hacker sẽ chiếm quyền `root` của container và có thể khai thác để tấn công thẳng vào máy host OS.
* **Giải pháp**: Khai báo một user thường (non-root) trong Dockerfile để chạy ứng dụng:
  ```dockerfile
  RUN useradd --create-home --uid 10001 appuser
  USER appuser
  ```
  Lúc này ứng dụng chạy dưới quyền hạn chế, hacker không thể cài đặt thêm phần mềm độc hại hay truy cập file hệ thống nhạy cảm.

---

## VI. Câu hỏi ôn tập tự đánh giá
Sếp hãy tự trả lời các câu hỏi sau để củng cố kiến thức:
1. **Câu 1**: Tại sao trong file cấu hình, ta không gán giá trị mặc định cho API Key nhạy cảm? Cơ chế "fail-fast" giúp ích gì khi deploy lên Render/Railway?
2. **Câu 2**: Tại sao Liveness Probe (/health) không được kết nối đến Redis, trong khi Readiness Probe (/ready) thì bắt buộc phải kiểm tra Redis?
3. **Câu 3**: Timing Attack là gì? Tại sao việc so sánh API Key bằng `==` lại bị Timing Attack, còn `secrets.compare_digest()` thì không?
4. **Câu 4**: Phân biệt cơ chế Rate Limiting bằng Fixed Window và Sliding Window. Tại sao Sliding Window lại an toàn hơn?
5. **Câu 5**: Luồng xử lý chi tiết của tín hiệu SIGTERM đối với Graceful Shutdown là gì? Tại sao phải gọi lại signal handler cũ của uvicorn?

---
*Tài liệu học tập được tổng hợp tự động bởi Antigravity AI Assistant.*
