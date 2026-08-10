# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn câu trả lời bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Tuấn Anh  Mã học viên: 2A202601669

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi triển khai ứng dụng lên Render hoặc Railway, nếu chúng ta quên cấu hình biến môi trường `AGENT_API_KEY` nhưng code lại để mặc định là `"changeme"`, ứng dụng vẫn khởi động thành công và chạy bình thường. Hậu quả là bất kỳ ai trên internet cũng có thể mò ra API Key mặc định `"changeme"` này để gọi API của ta miễn phí, hoặc ứng dụng sẽ chạy nhưng liên tục báo lỗi xác thực không hợp lệ làm nghẽn luồng. Việc "chết sớm" (Fail-fast) sẽ ném ra lỗi `ValidationError` ngay lúc deploy, ngăn ứng dụng khởi động và báo đỏ trực tiếp trên dashboard của platform, giúp ta phát hiện và bổ sung key nhạy cảm lập tức trước khi ứng dụng chạy công khai.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T10:30:00.123456+00:00", "user_id": "sv01", "tokens_in": 15, "tokens_out": 30, "cost_usd": 0.0009}`

Hai việc làm được với dòng log JSON này:
1. **Thống kê chi phí tự động**: Dùng các công cụ phân tích log tập trung (như Grafana Loki, Datadog) để tính tổng số tiền một người dùng (user_id) đã tiêu dùng trong ngày/tháng bằng cách cộng dồn trường `cost_usd`.
2. **Cảnh báo lỗi hệ thống**: Cấu hình bộ lọc đếm số lượng event có `level: "error"` để tự động gửi thông báo khẩn cấp qua Slack/Telegram khi tỷ lệ lỗi vượt ngưỡng cho phép (ví dụ: >5% số request trong 1 phút).

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) chính là toàn bộ môi trường và công cụ biên dịch thừa (như compiler gcc, build-essential, python header files, package cache, apt cache) được cài đặt ở stage `builder` để phục vụ việc build các python packages nhạy cảm. Ở bản Multi-stage, chúng ta đã bỏ qua các công cụ build nặng nề này ở stage runtime và chỉ sao chép các thư viện đã compile hoàn chỉnh sang stage sau, giúp image thu gọn tối đa.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

* Khi sửa code trong `app/main.py`, các layer cài đặt thư viện (như apt-get, COPY requirements.txt, RUN pip install) đều được dùng lại từ cache. Chỉ có layer `COPY . .` và các layer gán quyền `chown` hoặc chạy lệnh CMD phía sau là phải chạy lại.
* Nếu đặt `COPY . .` lên trước `RUN pip install`, chỉ cần sửa một ký tự trong code thì Docker sẽ lập tức vô hiệu hóa cache của layer `COPY . .` và toàn bộ các layer phía sau. Điều này bắt buộc Docker phải tải lại và cài đặt lại toàn bộ các thư viện Python từ đầu, làm thời gian build tăng từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

* **Chuỗi sự kiện**: Code Python của ứng dụng có lỗ hổng bảo mật (ví dụ: RCE - Remote Code Execution). Hacker khai thác lỗ hổng này để chạy các dòng lệnh shell tùy ý. Vì container chạy mặc định bằng user `root`, các dòng lệnh độc hại này cũng sẽ chạy dưới quyền root trong container. Hacker có thể tận dụng các lỗ hổng nhân hệ điều hành để thực hiện container escape (thoát khỏi container), trực tiếp chiếm đoạt quyền root điều khiển máy host vật lý.
* **Lệnh USER**: Lệnh `USER appuser` chuyển tiến trình chạy ứng dụng sang user thường (non-root). Khi hacker khai thác thành công RCE, họ chỉ có quyền hạn cực kỳ hạn chế của `appuser`. Họ không thể cài đặt thêm mã độc, không thể đọc file hệ thống nhạy cảm của container và không thể leo thang đặc quyền để thoát ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

* Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
* **Giải thích**: Người dùng gửi 10 request vào giây `10:00:59` (giây cuối cùng của phút trước) và gửi tiếp 10 request nữa vào giây `10:01:01` (giây đầu tiên của phút sau). Hệ thống đếm theo phút đồng hồ sẽ reset bộ đếm về 0 vào thời khắc `10:01:00`, do đó cả hai đợt gửi đều được tính là hợp lệ (không quá 10/phút), nhưng thực tế máy chủ phải hứng chịu 20 request dồn dập chỉ trong 2 giây.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

* **Khác nhau**: Rate limit giới hạn **số lượng cuộc gọi (request)** của người dùng trong một khoảng thời gian. Cost guard giới hạn **số tiền chi tiêu (hoặc token)** của người dùng để bảo vệ ngân sách.
* **Tình huống 1 (Rate limit cho qua nhưng Cost guard chặn)**: User chỉ gọi 1 request trong vòng 1 phút (đúng hạn mức 10/phút). Tuy nhiên request này tải lên một file tài liệu khổng lồ 100,000 tokens làm chi phí ước tính vượt quá ngân sách tháng còn lại của họ.
* **Tình huống 2 (Ngược lại)**: User gửi 20 request cực ngắn liên tục (mỗi câu chỉ 1 từ "Hi", tốn 0.00001 USD) trong 5 giây. Chi phí cực nhỏ không đáng kể (Cost guard cho qua) nhưng tốc độ gọi quá nhanh gây nghẽn băng thông server (Rate limit phải chặn để chống DDoS).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis gặp sự cố mất kết nối trong 30 giây.
2. Cả 3 container chạy endpoint gộp chung kiểm tra Redis và đồng loạt báo lỗi `503 Service Unavailable`.
3. Hệ thống quản lý (orchestrator) nhận thấy cả 3 container báo lỗi liveness probe nên lập tức ra lệnh kill và khởi động lại (restart) cả 3 container cùng lúc.
4. Trong lúc khởi động lại, các container vẫn cố gắng ping Redis và tiếp tục báo lỗi unhealthy, dẫn đến việc bị reboot liên tục (vòng lặp Restart Loop).
5. Khi Redis hoạt động bình thường trở lại, hệ thống vẫn bị sập hoàn toàn và mất nhiều thời gian để phục hồi thay vì tự động hoạt động lại ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần with cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lịch sử lưu bằng dict trong bộ nhớ RAM (stateful), giá trị `history_length` sẽ nhảy loạn xạ hoặc reset liên tục (ví dụ: đang 3 nhảy xuống 1 rồi lên 2) tùy thuộc vào việc Load Balancer định tuyến request rơi trúng container nào (container A, B hay C). Người dùng sẽ thấy AI bị mất ngữ cảnh trò chuyện liên tục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

* **Lỗi**: Ứng dụng deploy lên Render thành công nhưng cổng `/ready` luôn báo lỗi `503 Service Unavailable` và không nhận được traffic (Readiness probe failed).
* **Tìm nguyên nhân**: Mở tab logs của Web Service trên Render, thấy báo lỗi kết nối Redis bị từ chối (`Connection refused`).
* **Cách sửa**: Copy địa chỉ kết nối nội bộ (Internal Connection String) của Redis Service trên dashboard Render và dán đè vào biến môi trường `REDIS_URL` của Web Service, sau đó tiến hành redeploy lại.
