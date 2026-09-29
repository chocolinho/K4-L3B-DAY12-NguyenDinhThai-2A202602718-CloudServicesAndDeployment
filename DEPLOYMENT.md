# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Đình Thái |
| Mã học viên | 2A202602718 |
| Repo | https://github.com/chocolinho/K4-L3B-DAY12-NguyenDinhThai-2A202602718-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-6f3a.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Railway Redis add-on (${{Redis.REDIS_URL}}) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

### Dành cho Windows PowerShell:

```powershell
$URL = "https://day12-agent-production-6f3a.up.railway.app"
$KEY = $env:AGENT_API_KEY  # Hoặc thay bằng giá trị API Key của bạn nếu chưa set biến môi trường

# 1. Liveness — mong đợi 200 {"status":"ok"}
curl.exe -i "$URL/health"

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl.exe -i "$URL/ready"

# 3. Không có API key — mong đợi 401
curl.exe -i -X POST "$URL/ask" `
  -H "Content-Type: application/json" `
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
# Lưu ý trong PowerShell: dùng nháy đơn bao bọc chuỗi JSON '{"question":"..."}', KHÔNG escape '{\"question\":...}' để tránh lỗi 422
curl.exe -i -X POST "$URL/ask" `
  -H "Content-Type: application/json" `
  -H "X-API-Key: $KEY" `
  -H "X-User-Id: sv-test" `
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, các lần vượt hạn mức trả về 429
1..15 | ForEach-Object {
    curl.exe -s -o NUL -w "%{http_code} " `
      -X POST "$URL/ask" `
      -H "Content-Type: application/json" `
      -H "X-API-Key: $KEY" `
      -H "X-User-Id: sv-test" `
      -d '{"question":"test"}'
}
Write-Host
```

### Dành cho Bash (Linux/macOS):

```bash
URL="https://day12-agent-production-6f3a.up.railway.app"

# 1. Liveness
curl -i $URL/health

# 2. Readiness
curl -i $URL/ready

# 3. Không có API key
curl -i -X POST $URL/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'

# 4. Có API key
curl -i -X POST $URL/ask -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{"question":"Deploy là gì?"}'

# 5. Rate limit
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $URL/ask -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness probe:
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness probe:
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ready","redis":true}

# 3. Request không có API key:
HTTP/1.1 401 Unauthorized
Content-Type: application/json
{"detail":"invalid or missing API key"}

# 4. Request có API key:
HTTP/1.1 200 OK
Content-Type: application/json
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

# 5. Rate limit 15 request liên tiếp:
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
