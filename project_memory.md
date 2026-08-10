# Project Memory - K3-Day12-And-Deployment

Dự án này tập trung vào việc cấu trúc hạ tầng cloud và triển khai (deployment) cho một AI Agent viết bằng FastAPI, tuân thủ các tiêu chuẩn bảo mật, tối ưu hóa tài nguyên (Docker multi-stage), quản lý chi phí và đảm bảo độ tin cậy/sẵn sàng (scaling & reliability).

## Tech Stack
- **Backend**: Python 3.11+, FastAPI, Uvicorn
- **Database/Caching**: Redis (quản lý rate limiting, hội thoại stateless)
- **Containerization**: Docker, Docker Compose, Nginx (Load Balancer)
- **Deployment Cloud Platforms**: Railway, Render
- **Testing**: Pytest

## Project Structure
- `app/`: Thư mục mã nguồn chính của API Agent.
  - `config.py`: Quản lý cấu hình theo chuẩn 12-Factor (Pydantic Settings).
  - `logging_utils.py`: Cấu hình log định dạng JSON phục vụ cho việc thu thập log tập trung.
  - `main.py`: Entrypoint của FastAPI app, khai báo các endpoints và middleware.
  - `auth.py`: Module xác thực qua API Key.
  - `rate_limiter.py`: Triển khai sliding window rate limit bằng Redis.
  - `cost_guard.py`: Quản lý và giới hạn ngân sách gọi LLM theo tháng.
  - `store.py`: Lưu trữ lịch sử hội thoại dạng stateless sử dụng Redis.
  - `lifecycle.py`: Xử lý khởi tạo và giải phóng tài nguyên (lifecycle events) và tắt ứng dụng an toàn (graceful shutdown).
- `nginx/`: Nginx cấu hình làm Load Balancer.
- `tests/`: Các bài test tự động cho từng checkpoint (CP1 - CP5).
- `Dockerfile`: Cấu hình build Docker image dạng multi-stage tối ưu dung lượng và bảo mật.
- `docker-compose.yml`: Cấu hình chạy ứng dụng cục bộ kèm Redis và Nginx.
- `exercises.md`: Tài liệu phản ánh 10 câu hỏi lý thuyết/thực tế về Deployment & Cloud Infrastructure.
- `DEPLOYMENT.md`: Điền URL môi trường production sau khi deploy thành công.

## Tiến Độ Phát Triển

- **2026-08-10**:
  * Đã hoàn tất 100% mã nguồn cho tất cả 5 checkpoints chính (CP1 -> CP5).
  * Đã deploy thành công ứng dụng thật lên Railway với Public URL hoạt động: `https://agent-production-e157.up.railway.app`.
  * Kết nối thành công cơ sở dữ liệu Redis thật trên cloud.
  * Bộ test kiểm thử CP5 thật qua Internet đã PASS 100%.
  * Phần lý thuyết Exercises đạt 10/10 câu tối đa.
  * Tổng điểm đạt **100.0/100** điểm trần của bài Lab K3-Day12.
  * Dự án đã ở trạng thái hoàn thiện tuyệt đối và sẵn sàng nộp bài.
