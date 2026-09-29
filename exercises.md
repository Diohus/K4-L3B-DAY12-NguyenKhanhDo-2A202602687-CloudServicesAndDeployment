# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng đáp án mẫu dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Khánh Đô.  Mã học viên: 2A202602687.

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu tôi quên đặt `AGENT_API_KEY` khi tạo service Railway, lúc khởi tạo `Settings` sẽ báo thiếu trường bắt buộc. Nhờ vậy tôi có thể thấy lỗi cấu hình và sửa biến môi trường trước khi đưa URL cho người khác dùng. Nếu để mặc định `changeme`, service có thể vẫn chạy; người biết khóa mặc định có thể gọi `/ask` và sử dụng quota của tôi mà tôi không nhận ra lúc deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log quan sát được khi gọi service local ngày 29/09/2026:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:49:16.377449+00:00", "user_id": "codex-smoke", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

> Từ dòng JSON trên, tôi có thể lọc các lượt gọi của `user_id=codex-smoke` theo thời gian, rồi cộng `cost_usd` để biết người dùng đó đã tiêu bao nhiêu. Tôi cũng có thể cộng `tokens_in` và `tokens_out` để theo dõi mức sử dụng token. Dòng `print("đã trả lời xong")` không có các trường này nên không hỗ trợ hai phép thống kê đó.

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
| 1 stage (bản đầu) | 1.73 GB (Docker báo) |
| Multi-stage | 271 MB (Docker báo) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản đầu dùng `python:3.11` đầy đủ, cài thư viện ngay trong image cuối và không dùng `--no-cache-dir`, nên mang theo base image lớn cùng dữ liệu cài đặt của pip. Bản mới dùng `python:3.11-slim`; dependencies được cài ở stage builder rồi chỉ copy kết quả sang runtime. Builder không nằm trong image cuối. Docker báo bản đầu 1,73 GB và bản mới 271 MB, chênh khoảng 1,46 GB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Kiểm chứng thực tế: thêm tạm một file vào `app/` rồi build lại. Docker báo
`COPY requirements.txt` và `RUN pip install` là `CACHED`; `COPY app ./app` và
`COPY utils ./utils` chạy lại. File thử đã được xóa sau khi build.

> Khi nội dung `app/` đổi, Docker vẫn dùng cache cho `COPY requirements.txt` và `RUN pip install`; lần kiểm chứng của tôi, hai bước đó đều hiện `CACHED`. `COPY app ./app` chạy lại, và `COPY utils ./utils` cũng chạy lại vì nó nằm sau layer thay đổi. Nếu `COPY . .` đứng trước `RUN pip install`, thay đổi source sẽ làm mất cache của bước cài thư viện, khiến pip chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng cho phép chạy lệnh trong app có thể cho kẻ tấn công quyền của tiến trình bên trong container. Nếu tiến trình chạy bằng root và container còn có mount hoặc cấu hình thiếu an toàn, quyền đó có thể được dùng để sửa dữ liệu trên host hoặc kết hợp với lỗ hổng thoát container. `USER appuser` khiến tiến trình chỉ có quyền của user thường trong container, giảm quyền có được ngay sau khi khai thác. Nó không tự bảo đảm host an toàn nếu Docker hoặc mount vẫn cấu hình sai.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi 20 request trong 2 giây: gửi 10 request lúc 10:00:59, rồi thêm 10 request lúc 10:01:01. Bộ đếm theo phút coi đây là hai phút khác nhau nên cho qua cả hai nhóm. Sliding window 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau đến, nên sẽ chặn sau khi đủ 10 request trong 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số request trong 60 giây; cost guard giới hạn tổng USD của từng user trong tháng. Ví dụ một user chỉ gửi 1 request trong phút này nhưng tổng chi phí tháng đã vượt 10 USD: rate limit cho qua, cost guard trả 402. Ngược lại, một user mới tiêu rất ít tiền nhưng gửi request thứ 11 trong vòng 60 giây: cost guard vẫn còn ngân sách, rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối thì cả ba container đều kiểm tra Redis ở endpoint gộp và đồng loạt trả 503. Nếu orchestrator dùng endpoint đó làm liveness probe, nó sẽ kết luận cả ba process bị hỏng và restart chúng. Trong thời gian restart, cụm không còn instance phục vụ; khi Redis trở lại, các container còn phải khởi động lại rồi mới nhận traffic. Tách `/health` chỉ kiểm tra process và `/ready` kiểm tra Redis giúp load balancer tạm ngừng gửi request khi Redis lỗi mà không restart cả cụm.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Quan sát local với cùng `X-User-Id`: hai request liên tiếp cho
`history_length=0` rồi `history_length=2`.

> Tôi đã xác minh trên stack local rằng hai request cùng `X-User-Id` cho `history_length` lần lượt là 0 và 2. Bài test dùng hai `ConversationStore` khác nhau trên cùng Redis cũng cho thấy instance thứ hai đọc được message do instance thứ nhất ghi. Tôi chưa đo trực tiếp ba container sau load balancer. Nếu dùng dict riêng trong mỗi process, mỗi container chỉ thấy các lượt nó từng xử lý; khi request chuyển sang container khác, `history_length` có thể giảm hoặc quay về 0 thay vì tăng đều 0, 2, 4, ...

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Sự cố đã quan sát khi kiểm tra Railway: lệnh `curl` dùng hostname không có
`https://` nhận HTTP 301 với header `Location` dẫn đến URL HTTPS. Sau khi thêm
`https://` vào `$url`, `/health` và `/ready` trả 200, còn `/ask` không có key
trả 401.

> Khi kiểm tra bản Railway, tôi đặt `$url` chỉ bằng hostname. `curl.exe` trả `HTTP/1.1 301 Moved Permanently` cho cả ba endpoint, kèm header `Location: https://day12-agent-production-52c1.up.railway.app/...`. Header đó cho thấy tôi đang gọi HTTP và Railway chuyển sang HTTPS. Tôi sửa `$url` thành URL bắt đầu bằng `https://`; sau đó `/health` và `/ready` trả 200, còn `/ask` không có key trả 401 như dự kiến. Đây là lỗi trong bước kiểm tra URL sau deploy, không phải lỗi build của service.
