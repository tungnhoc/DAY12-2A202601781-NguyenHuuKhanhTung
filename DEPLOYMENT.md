# Thông Tin Deploy — Checkpoint 5

> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Hữu Khánh Tùng |
| Mã học viên | 2A202601781 |
| Repo | https://github.com/tungnhoc/DAY12-2A202601781-NguyenHuuKhanhTung |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-x2ty.onrender.com |
| Platform | Render Blueprint (Docker runtime + Redis) |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự gán tự động |
| `AGENT_API_KEY` | ✅ | đặt trong Render Blueprint Environment |
| `REDIS_URL` | ✅ | trỏ tự động tới service `day12-redis` qua `fromService` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-x2ty.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-x2ty.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-x2ty.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-x2ty.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên:

```json
HTTP/1.1 200 OK
content-type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}

HTTP/1.1 200 OK
content-type: application/json

{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
content-type: application/json

{"detail":"invalid or missing API key"}

HTTP/1.1 200 OK
content-type: application/json

{"answer":"Ngắn gọn: Hello phụ thuộc vào ba yếu tố...","user_id":"sv-test","history_length":2,"cost_usd":3.255e-05,"tokens":{"in":37,"out":45}}
```

## Ảnh Chụp Màn Hình

Ảnh đặt tại thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý Render Blueprint dịch vụ day12-agent và day12-redis.
- `screenshots/health.png` — kết quả gọi /health trực tiếp từ live domain.
- `screenshots/ready.png` — kết quả gọi /ready trực tiếp từ live domain.
- `screenshots/ask.png` — kết quả gọi /ask không truyền API key (401 Unauthorized).
- `screenshots/ask(có key).png` — kết quả gọi /ask truyền API key live thành công (200 OK).

