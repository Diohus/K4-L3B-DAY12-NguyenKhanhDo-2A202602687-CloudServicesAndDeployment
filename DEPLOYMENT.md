# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyen Khanh Do (theo tên repository; kiểm tra lại dấu trước khi nộp) |
| Mã học viên | 2A202602687 |
| Repo | https://github.com/Diohus/K4-L3B-DAY12-NguyenKhanhDo-2A202602687-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-52c1.up.railway.app |
| Platform | Railway |
| Ngày xác minh | 29/09/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Railway Redis; `/ready` xác nhận kết nối hoạt động |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Chạy các lệnh sau trong **PowerShell**, tại thư mục gốc repo. Khóa API được đọc
từ `.env` vào biến trong phiên terminal, không in ra màn hình. Các lệnh giả định
khóa local giống khóa đã đặt trên Railway; nếu khác, dùng khóa Railway cho biến
`$apiKey` trong phiên PowerShell (không ghi vào tài liệu hoặc commit).

```powershell
$url = "https://day12-agent-production-52c1.up.railway.app"
$apiKey = ((Get-Content .env | Where-Object { $_ -match '^AGENT_API_KEY=' } | Select-Object -First 1) -replace '^AGENT_API_KEY=', '')
$body = @{ question = 'Hello' } | ConvertTo-Json -Compress

# 1–2. Liveness và readiness: mong đợi status=ok và status=ready
Invoke-RestMethod -Uri "$url/health"
Invoke-RestMethod -Uri "$url/ready"

# 3. Thiếu key: PowerShell ném exception cho HTTP 401; in mã trạng thái
try {
    Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$url/ask" -ContentType 'application/json' -Body $body | Out-Null
} catch {
    [int]$_.Exception.Response.StatusCode
}

# 4. Có key: mong đợi 200 và câu trả lời
$headers = @{ 'X-API-Key' = $apiKey; 'X-User-Id' = 'sv-ps-check' }
$response = Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$url/ask" -ContentType 'application/json' -Headers $headers -Body $body
$response.StatusCode
$response.Content

# 5. Rate limit: user riêng để không bị ảnh hưởng bởi các lần thử trước
$rateHeaders = @{ 'X-API-Key' = $apiKey; 'X-User-Id' = "rate-$(Get-Date -Format yyyyMMddHHmmss)" }
for ($i = 1; $i -le 15; $i++) {
    try {
        (Invoke-WebRequest -UseBasicParsing -Method Post -Uri "$url/ask" -ContentType 'application/json' -Headers $rateHeaders -Body $body).StatusCode
    } catch {
        [int]$_.Exception.Response.StatusCode
    }
}
```

## Kết Quả Chạy Thật

Kết quả gọi service công khai qua HTTPS ngày 29/09/2026. Giá trị API key được
đọc từ `.env` local để kiểm tra và không được in vào tài liệu:

```text
GET /health 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET /ready 200 {"status":"ready","redis":true}
POST /ask 401 {"detail":"invalid or missing API key"}
POST /ask với X-API-Key hợp lệ 200, có trường answer
15 lần POST /ask cùng X-User-Id: 200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

`pytest tests/test_cp5.py` qua 8 bài, bỏ qua bài xác thực khi không đặt
`DEPLOY_API_KEY`. Khi truyền `DEPLOY_API_KEY` cùng giá trị key đã xác minh,
checkpoint qua 9 bài; 4 bài fallback được bỏ qua do đang dùng cloud.

### Kiểm tra local ngày 29/09/2026

Stack Docker Compose đã chạy với `agent` và `redis` ở trạng thái healthy. Sau khi
dựng lại `agent` với cấu hình `.env` hiện tại, các lời gọi đến
`http://localhost:8000` cho kết quả:

```text
GET  /health                         200
GET  /ready                          200
POST /ask không có X-API-Key         401
POST /ask có X-API-Key hợp lệ        200, history_length=0
POST /ask lần tiếp theo cùng user    200, history_length=2
```

Đây là kết quả local, được ghi riêng để đối chiếu với kết quả Railway phía trên.

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
