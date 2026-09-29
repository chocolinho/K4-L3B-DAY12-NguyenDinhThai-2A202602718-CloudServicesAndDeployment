# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng *Câu trả lời của bạn* bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Đình Thái  Mã học viên: 2A202602718

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: khi deploy lên Railway, mình quên set biến `AGENT_API_KEY` trong dashboard. Vì trường `agent_api_key` trong `Settings` không có giá trị mặc định, pydantic-settings raise `ValidationError` ngay khi app khởi động → container restart liên tục → mình thấy log lỗi ngay trên dashboard và biết phải thêm biến. Nếu để mặc định `"changeme"`, app sẽ khởi động bình thường, endpoint `/ask` vẫn hoạt động nhưng bất kỳ ai biết khóa mặc định đều gọi được API, tiêu tiền token của mình mà mình không hay biết cho đến khi nhận hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T05:05:49.123456+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được:

1. **Lọc và truy vấn tự động:** Trên Railway/Datadog, mình có thể query `event == "ask_completed" AND cost_usd > 0.01` để tìm các request tốn kém bất thường. Với `print()` chỉ có chuỗi text thuần, không có trường nào để máy phân tách và lọc.

2. **Thống kê và cảnh báo:** Mình có thể tổng hợp trường `cost_usd` theo `user_id` để biết ai đang tiêu nhiều nhất trong tháng, hoặc đặt alert khi `tokens_out` vượt ngưỡng. Với `print()` thì phải parse regex thủ công và rất dễ sai khi format thay đổi.

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
| 1 stage (bản đầu) | ~420 MB |
| Multi-stage | ~180 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch (~240 MB) chủ yếu là các công cụ build (gcc, pip cache, header files C, wheel metadata) được dùng trong quá trình `pip install` để compile các thư viện có phần mở rộng C (như `pydantic-core`, `redis`). Ở bản 1 stage, tất cả đều nằm trong cùng image cuối. Với multi-stage, stage `builder` chứa toàn bộ công cụ build này nhưng stage `runtime` chỉ copy kết quả đã cài xong (`/install`) sang, nên các công cụ build không còn xuất hiện trong image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Với Dockerfile hiện tại, thứ tự là `COPY requirements.txt` → `RUN pip install` → `COPY app ./app`. Khi chỉ sửa 1 ký tự trong `app/main.py`:
- **Dùng lại cache:** `COPY requirements.txt .` và `RUN pip install` (vì `requirements.txt` không đổi).
- **Phải chạy lại:** `COPY app ./app` và các layer sau nó (vì nội dung thư mục `app/` đã thay đổi).

Nếu đặt `COPY . .` lên **trước** `RUN pip install`:
- Bất kỳ thay đổi nào trong bất kỳ file nào (kể cả 1 ký tự trong `main.py`) sẽ làm cache của `COPY . .` bị vô hiệu → `RUN pip install` cũng phải chạy lại từ đầu, mất thêm 30–60 giây mỗi lần build. Đây là lý do tách `COPY requirements.txt` ra trước: dependency ít khi thay đổi, còn source code thay đổi liên tục.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Code Python có lỗ hổng (ví dụ: endpoint `/ask` nhận input không sanitize, cho phép Server-Side Template Injection hoặc OS command injection).
2. Kẻ tấn công khai thác lỗ hổng để chạy lệnh tùy ý bên trong container.
3. Vì process chạy bằng **root** (UID 0), lệnh của kẻ tấn công cũng có quyền root trong container.
4. Với quyền root, kẻ tấn công có thể mount filesystem của host, đọc `/proc`, khai thác các lỗ hổng kernel escape (như CVE cũ của runc) để thoát ra ngoài container.
5. Khi thoát ra host với quyền root, kẻ tấn công kiểm soát toàn bộ máy.

Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi ở **bước 3**: dù kẻ tấn công chạy được lệnh trong container, lệnh đó chỉ có quyền của user thường — không thể mount filesystem, không thể ghi vào thư mục hệ thống, và các kỹ thuật escape kernel hầu hết đều yêu cầu UID 0.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** trong 2 giây liên tiếp.

Cách đạt được: gửi 10 request vào lúc 10:00:**59** (cuối phút 10:00, vẫn hợp lệ vì phút đó mới có 10 request) → đồng hồ sang 10:01:**00**, bộ đếm reset về 0 → gửi thêm 10 request vào lúc 10:01:**01** (đầu phút mới, bộ đếm mới chỉ có 10). Tổng cộng 20 request trong khoảng 10:00:59 – 10:01:01, tức 2 giây, gấp đôi hạn mức mong muốn.

Sliding window 60 giây không có lỗ hổng này vì nó luôn nhìn lại đúng 60 giây gần nhất: tại thời điểm 10:01:01, cửa sổ bao gồm cả 10 request lúc 10:00:59, nên bộ đếm hiện tại là 10, request thứ 11 sẽ bị chặn ngay.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Khác nhau cơ bản:** Rate limit giới hạn **tần suất** (số request / phút), còn cost guard giới hạn **chi phí tích lũy** (tổng tiền USD / tháng).

**Tình huống rate limit cho qua, cost guard chặn:** Một user gửi đều đặn 9 request/phút (dưới hạn mức 10), nhưng mỗi request hỏi câu rất dài với 50.000 token. Sau vài ngày, tổng chi phí tích lũy vượt $10/tháng → cost guard trả 402, dù rate limit vẫn cho qua vì tần suất hợp lệ.

**Tình huống cost guard cho qua, rate limit chặn:** Đầu tháng, user mới chỉ gửi 2 request nhỏ (tổng chi phí $0.001, rất xa ngân sách $10). Nhưng ngay sau đó user gửi liên tục 15 request trong 1 phút → rate limit chặn ở request thứ 11 (trả 429), trong khi cost guard vẫn thấy ngân sách còn dư rất nhiều.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. **t=0s:** Redis mất kết nối (do restart, network blip, v.v.).
2. **t=0–5s:** Orchestrator (Docker/Railway/K8s) gọi endpoint health gộp, endpoint kiểm tra Redis → thất bại → trả 503.
3. **t=5–15s:** Health check fail liên tiếp vài lần (tùy `retries`), orchestrator kết luận container "chết" → **restart cả 3 container**.
4. **t=15–25s:** Cả 3 container đang restart, không có container nào sẵn sàng nhận request → **toàn bộ service ngừng hoạt động** (downtime 100%).
5. **t=25–30s:** Redis quay lại, nhưng 3 container vẫn đang trong quá trình khởi động lại → user vẫn thấy 502/503.
6. **t=30s+:** Container khởi động xong, Redis đã sẵn sàng → service phục hồi.

Nếu tách riêng: `/health` (liveness, không check Redis) vẫn trả 200 → orchestrator không restart container. `/ready` (readiness, check Redis) trả 503 → load balancer chỉ **ngừng đẩy traffic mới** vào, không restart. Khi Redis quay lại sau 30s, `/ready` trả 200 trở lại → traffic tự phục hồi, **không có downtime**.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis (hiện tại): `history_length` tăng đều đặn 0, 2, 4, 6, 8... bất kể request rơi vào instance nào, vì cả 3 instance đều đọc/ghi cùng một Redis list `history:sv-test`.

Nếu dùng dict Python trong RAM: `history_length` sẽ **nhảy lung tung**, ví dụ: 0, 0, 0, 2, 0, 2, 4, 2... Lý do: Nginx/load balancer phân phối request round-robin vào 3 instance. Request 1 vào instance A (ghi 2 message vào dict A), request 2 vào instance B (dict B rỗng → `history_length` = 0, agent "mất trí nhớ"), request 3 vào instance C (cũng rỗng). User sẽ thấy agent liên tục quên cuộc hội thoại. Ngoài ra, khi container bị restart (deploy mới, scale down), toàn bộ dict trong RAM mất sạch — lịch sử biến mất vĩnh viễn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Lỗi:** Sau khi deploy lên Railway, endpoint `/ready` trả về `500 Internal Server Error` thay vì `200`. Khi kiểm tra log trên Railway dashboard, thấy:
```
ValueError: Redis URL must be a valid URL starting with redis://
```

**Nguyên nhân:** Khi set biến `REDIS_URL` trên Railway dashboard, mình đã đặt giá trị trong dấu ngoặc kép: `REDIS_URL="${{Redis.REDIS_URL}}"`. Railway lưu nguyên cả dấu `"` vào giá trị, khiến URL trở thành `"redis://default:pass@host:port"` (có dấu ngoặc kép bao ngoài) → thư viện `redis-py` không parse được.

**Cách tìm ra:** Đọc log trên Railway dashboard, thấy stack trace chỉ vào `redis.from_url()` với giá trị URL không hợp lệ. So sánh với giá trị raw trong dashboard Variables thì thấy dấu ngoặc kép thừa.

**Cách sửa:** Xóa biến cũ, tạo lại `REDIS_URL` với giá trị raw không có dấu ngoặc kép: `${{Redis.REDIS_URL}}`. Sau khi redeploy, `/ready` trả `200 {"status":"ready","redis":true}`.

