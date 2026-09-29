# Thông Tin Deploy — Checkpoint 5
## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Hoàng Quân |
| Mã học viên | 2A202603016 |
| Repo | https://github.com/The-Spirit-of-the-Beehive/K4-L3B-DAY12-DoHoangQuan-2A202603016-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-xlv0.onrender.com |
| Platform | Render |
| Ngày deploy | 29/9/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Platform tự gán |
| `AGENT_API_KEY` | ✅ | Đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value / Redis Service |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra
```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-xlv0.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-xlv0.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-xlv0.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent-xlv0.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-xlv0.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 05:58:57 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: 19ad3b1f-945d-4b2b
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a428c24ce9f5dd8d-HKG
alt-svc: h3=":443"; ma=86400

{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 06:27:54 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: 6a871067-8161-4a15
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a428ec6ccbdd04b9-HKG
alt-svc: h3=":443"; ma=86400

{"status":"ready","redis":true}

#3. Không có API key — mong đợi 401
HTTP/1.1 401 Unauthorized
Date: Tue, 29 Sep 2026 06:28:45 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
cf-cache-status: DYNAMIC
rndr-id: c2ea1686-1c2a-44ba
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
CF-RAY: a428edf6cc76848d-HKG
alt-svc: h3=":443"; ma=86400

{"detail":"invalid or missing API key"}

# 4. Có API key — mong đợi 200 kèm câu trả lời
HTTP/1.1 200 OK
Date: Tue, 29 Sep 2026 06:38:11 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
rndr-id: 4d58ed80-b199-49b5
Server: cloudflare
vary: Accept-Encoding
x-render-origin-server: uvicorn
cf-cache-status: DYNAMIC
CF-RAY: a428fbc69b5b9b90-SIN
alt-svc: h3=":443"; ma=86400

{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên. (Mình đang nhớ 20 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":20,"cost_usd":9.405e-05,"tokens":{"in":439,"out":47}}

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

## Ảnh Chụp Màn Hình
- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl