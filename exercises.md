# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.

Họ và tên: Nguyễn Hữu Khánh Tùng  Mã học viên: 2A202601781

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu đặt giá trị mặc định `"changeme"`, ứng dụng khi deploy lên môi trường Production trên Render mà lập trình viên quên cấu hình biến môi trường `AGENT_API_KEY` vẫn sẽ khởi động thành công. Kẻ tấn công có thể dễ dàng đoán được khóa mặc định `"changeme"` và gọi tự do vào API `/ask`, làm cạn kiệt hạn mức LLM và gây thiệt hại tài chính lớn. Ngược lại, việc ứng dụng "chết sớm" (Fail Fast) ném lỗi `ValidationError` ngay lúc vừa khởi chạy sẽ buộc tiến trình Deploy trên Render dừng lại ngay và hiển thị lỗi trên Dashboard log, ngăn ngừa rò rỉ bảo mật trước khi bất kỳ traffic người dùng nào chạm vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")` không làm được.

> `{"timestamp": "2026-08-10T10:59:45Z", "event": "ask_completed", "level": "info", "user_id": "sv-test", "tokens_in": 37, "tokens_out": 45, "cost_usd": 0.00003255}`
> 
> **Hai việc làm được với log JSON**:
> 1. **Thu thập và thống kê tự động (Log Aggregation & Analytics)**: Đơn vị quản lý log (Datadog, ElasticSearch/Kibana, CloudWatch) có thể tự động bóc tách các trường JSON để tạo biểu đồ theo dõi tổng chi phí `cost_usd` và lượng token tiêu thụ theo từng `user_id` theo thời gian thực.
> 2. **Lọc và phát cảnh báo tự động (Structured Alerting)**: Có thể viết câu lệnh truy vấn lọc chính xác các log có `cost_usd > 0.05` hoặc `level == "error"` để gửi thông báo cảnh báo trực tiếp về Slack/Telegram. Hàm `print()` chỉ in text thô không cấu trúc nên máy tính không thể parse dữ liệu tự động được.

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
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~835 MB) chứa bộ công cụ biên dịch mã nguồn C/C++ (`gcc`, `g++`, `make`), bộ thư viện phát triển OS (`python3-dev`), bộ nhớ đệm wheel của pip (`~/.cache/pip`), và các tiện ích hệ điều hành không cần thiết trong base image tiêu chuẩn. Bản Multi-stage ở stage `runtime` chỉ dùng base image `python:3.11-slim` siêu nhẹ và chỉ copy kết quả từ stage `builder` sang, loại bỏ hoàn toàn các công cụ biên dịch thừa giúp kéo/đẩy image trên Render cực nhanh.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt `COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với cấu trúc Dockerfile hiện tại (`COPY requirements.txt .` -> `RUN pip install` -> `COPY . .`), layer `RUN pip install` được **dùng lại 100% từ cache** vì file `requirements.txt` không hề thay đổi. Chỉ có layer `COPY . .` và các bước sau đó là phải chạy lại, giúp thời gian build chỉ mất 1-2 giây.
> 
> Nếu đặt `COPY . .` lên trước `RUN pip install`: Khi sửa 1 ký tự trong `app/main.py`, cache của layer `COPY . .` bị vô hiệu hóa, kéo theo layer `RUN pip install` đứng sau cũng bị vô hiệu hóa cache và buộc phải tải, cài lại toàn bộ thư viện từ Internet từ đầu, khiến mỗi lần sửa code đều mất vài phút build trên Cloud.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> **Chuỗi sự kiện**: Hacker khai thác lỗ hổng Remote Code Execution (RCE) trong ứng dụng Python ➔ Mở shell điều khiển bên trong container dưới quyền `root` ➔ Khai thác lỗ hổng thoát khỏi container (container escape) hoặc thao tác trên các socket hệ thống ➔ Chiếm quyền kiểm soát máy Host với quyền `root` tối cao.
> 
> **Vị trí lệnh `USER` cắt đứt**: Lệnh `USER appuser` ép tiến trình Python chạy dưới quyền user thường (UID 10001). Khi hacker tấn công RCE thành công vào Python, chúng chỉ có quyền của `appuser` không có đặc quyền root, không thể chỉnh sửa file hệ thống container, không thể thực thi câu lệnh quản trị và bị chặn đứng hoàn toàn không thể leo thang đặc quyền ra máy Host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được con số đó.

> **Tối đa 20 request trong 2 giây liên tiếp.**
> 
> **Giải thích**: Với Fixed Window (reset vào giây 00 mỗi phút): Người dùng gửi 10 request vào giây `10:00:59` (nằm trong phút 10:00) và gửi tiếp 10 request vào giây `10:01:01` (nằm trong phút 10:01). Cả 2 lượt đều hợp lệ theo hạn mức 10 req/phút của từng phút riêng lẻ. Tuy nhiên trong khoảng thời gian 2 giây liên tục từ `10:00:59` đến `10:01:01`, hệ thống đã phải gánh tới 20 request (gấp đôi hạn mức cho phép). Thuật toán Sliding Window khắc phục triệt để bằng cách tính đúng tổng số request trong cửa sổ 60s liên tục lùi về trước.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Sự khác nhau**: Rate Limit giới hạn **số lượng request/tần suất gọi** trong khoảng thời gian ngắn (req/phút); Cost Guard giới hạn **tổng ngân sách chi tiêu/số lượng token LLM** tiêu thụ trong kỳ (USD/tháng).
> 
> - **Rate Limit cho qua nhưng Cost Guard chặn**: User mới gọi 1 request trong phút (thỏa hạn mức 1 req < 10 req/phút), nhưng prompt đính kèm tài liệu 100,000 token khiến chi phí của request đó là $15.00, vượt quá ngân sách tháng $10.00 ➔ Cost Guard ném lỗi HTTP 402 Payment Required.
> - **Cost Guard cho qua nhưng Rate Limit chặn**: User mới tiêu $0.50 trong ngân sách tháng $10.00, nhưng gửi liên tiếp 15 request cực ngắn trong vòng 5 giây ➔ Rate Limit ném lỗi HTTP 429 Too Many Requests ở request thứ 11.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm 3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> **Thứ tự sự kiện (Cascading Failure)**:
> 1. Giây 0: Redis bị đứt kết nối mạng trong 30 giây.
> 2. Giây 5: Orchestrator kiểm tra Liveness endpoint, thấy kết nối Redis lỗi nên đánh giá container bị hỏng.
> 3. Giây 10: Orchestrator tiến hành tiêu diệt (kill) và khởi động lại (restart) đồng loạt cả 3 container.
> 4. Giây 15-30: Các container mới khởi động lại kiểm tra Redis để sẵn sàng phục vụ, nhưng Redis vẫn chưa khôi phục ➔ Liveness tiếp tục báo lỗi ➔ Orchestrator lại tiếp tục kill và restart container (vòng lặp CrashLoopBackOff).
> 5. Giây 30+: Redis khôi phục, nhưng toàn bộ 3 container đều đang ở trạng thái bị kill/restarting đồng thời ➔ Hệ thống bị sập hoàn toàn (Downtime). Tách biệt `/ready` giúp Load Balancer chỉ ngừng nhận traffic mới mà KHÔNG restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một `X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Nếu lưu trong dict Python: Con số `history_length` sẽ thay đổi nhảy vọt thất thường (ví dụ: `0 -> 0 -> 2 -> 0 -> 4 -> 2...`) giữa các lần gọi API, do Load Balancer điều hướng mỗi request tới một container khác nhau và RAM của từng container chỉ giữ một phần lịch sử riêng lẻ.
> 
> Khi lưu trong Redis (Stateless): Giá trị `history_length` tăng đều đặn theo cấp số cộng (`0 -> 2 -> 4 -> 6 -> 8...`) ở mọi request bất kể được xử lý bởi container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Thông báo lỗi**: `redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379. Connection refused.` trên Render Deployment Log.
> 
> **Nguyên nhân**: Khi vừa khởi tạo trên Render, ứng dụng đọc biến mặc định `REDIS_URL=redis://localhost:6379/0` trong khi dịch vụ Redis trên Render chạy độc lập qua DNS `day12-redis`.
> 
> **Cách sửa**: Khai báo biến môi trường `REDIS_URL=${{Redis.REDIS_URL}}` (hoặc `fromService` trong `render.yaml`) để Render tự động trỏ đúng Connection URI của Redis Instance.
