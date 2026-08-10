# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Tuấn Anh |
| Mã học viên | 2A202601669 |
| Repo | https://github.com/nguyentuananh512005/DAY12-2A202601669-NguyenTuanAnh |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://agent-production-e157.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Railway Redis add-on connection string |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://agent-production-e157.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://agent-production-e157.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://agent-production-e157.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://agent-production-e157.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: my-super-secret-agent-api-key" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://agent-production-e157.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: my-super-secret-agent-api-key" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ok"}

HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
Content-Type: application/json
{"answer":"Deploy là quá trình đưa ứng dụng lên máy chủ...","user_id":"sv-test","history_length":0,"cost_usd":0.00015,"tokens":{"in":10,"out":20}}

200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
